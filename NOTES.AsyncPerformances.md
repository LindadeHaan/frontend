# Chapter 1: Asynchrony: Now & Later

## Review
A JavaScript program is (practically) always broken up into two or more chunks, where the first chunk runs *now* and the next chunk runs *later*, in response to an event. Even though the program is executed chunk-by-chunk, all of them share the same access to the program scope and state, so each modification to state is made on top of the previous state.

Whenever there are events to run, the event *loop* runs until the queue is empty. Each iteration of the event loop is a "tick." User interaction, IO, and timers enqueue events on the event queue.

At any given moment, only one event can be processed from the queue at a time. While an event is executing, it can directly or indirectly cause one or more subsequent events.

Concurrency is when two or more chains of events interleave over time, such that from a high-level perspective, they appear to be running *simultaneously* (even though at any given moment only one event is being processed).

It's often necessary to do some form of interaction coordination between these concurrent "processes" (as distinct from operating system processes), for instance to ensure ordering or to prevent "race conditions." These "processes" can also *cooperate* by breaking themselves into smaller chunks and to allow other "process" interleaving.

## A Program in Chunks
You may write your JS program in one *.js* file, but your program is almost certainly comprised of several chunks, only one of which is going to execute *now*, and the rest of which will execute *later*. The most common unit of chunk is the `function`.

The problem most developers new to JS seem to have is that *later* doesn't happen strictly and immediately after *now*. In other words, tasks that cannot complete *now* are, by definition, going to complete asynchronously, and thus we will not have blocking behavior as you might intuitively expect or want.

Consider:
```js
// ajax(..) is some arbitrary Ajax function given by a library
var data = ajax( "http://some.url.1" );

console.log( data );
// Oops! `data` generally won't have the Ajax results
```
You're probably aware that standard Ajax requests don't complete synchronously, which means the `ajax(..)` function does not yet have any value to return back to be assigned to `data` variable. If `ajax(..)` *could* block until the response came back, then the `data = ..` assignment would work fine.

But that's not how we do Ajax. We make an asynchronous Ajax request *now*, and we won't get the results back until *later*.

The simplest (but definitely not only, or necessarily even best!) way of "waiting" from *now* until *later* is to use a function, commonly called a callback function:
```js
// ajax(..) is some arbitrary Ajax function given by a library
ajax( "http://some.url.1", function myCallbackFunction(data){

	console.log( data ); // Yay, I gots me some `data`!

} );
```
Before you protest in disagreement, no, your desire to avoid the mess of callbacks is *not* justification for blocking, synchronous Ajax.

For example, consider this code:
```js
function now() {
	return 21;
}

function later() {
	answer = answer * 2;
	console.log( "Meaning of life:", answer );
}

var answer = now();

setTimeout( later, 1000 ); // Meaning of life: 42
```
There are two chunks to this program: the stuff that will run *now*, and the stuff that will run *later*. It should be fairly obvious what those two chunks are, but let's be super explicit:

Now:
```js
function now() {
	return 21;
}

function later() { .. }

var answer = now();

setTimeout( later, 1000 );
```
Later:
```js
answer = answer * 2;
console.log( "Meaning of life:", answer );
```
The *now* chunk runs right away, as soon as you execute your program. But `setTimeout(..)` also sets up an event (a timeout) to happen *later*, so the contents of the `later()` function will be executed at a later time (1,000 milliseconds from now).

Any time you wrap a portion of code into a `function` and specify that it should be executed in response to some event, you are creating a *later* chunk of your code, and thus introducing asynchrony to your program.

### Async Console
There is no specification or set of requirements around how the `console.*` methods work -- they are not officially part of JavaScript, but are instead added to JS by the *hosting environment*.

So, different browsers and JS environments do as they please, which can sometimes lead to confusing behavior.

In particular, there are some browsers and some conditions that `console.log(..)` does not actually immediately output what it's given. The main reason this may happen is because I/O is a very slow and blocking part of many programs (not just JS). So, it may perform better (from the page/UI perspective) for a browser to handle `console` I/O asynchronously in the background, without you perhaps even knowing that occurred.

A not terribly common, but possible, scenario where this could be observable (not from code itself but from the outside):
```js
var a = {
	index: 1
};

// later
console.log( a ); // ??

// even later
a.index++;
```
We'd normally expect to see the `a` object be snapshotted at the exact moment of the `console.log(..)` statement, printing something like `{ index: 1 }`, such that in the next statement when `a.index++` happens, it's modifying something different than, or just strictly after, the output of `a`.

Most of the time, the preceding code will probably produce an object representation in your developer tools' console that's what you'd expect. But it's possible this same code could run in a situation where the browser felt it needed to defer the console I/O to the background, in which case it's possible that by the time the object is represented in the browser console, the `a.index++` has already happened, and it shows `{ index: 2 }`.

It's a moving target under what conditions exactly `console` I/O will be deferred, or even whether it will be observable. Just be *aware* of this possible asynchronicity in I/O in case you ever run into issues in debugging where objects have been modified *after* a `console.log(..)` statement and yet you see the unexpected modifications show up.

## Event Loop
JavaScript itself has actually never had any direct notion of asynchrony built into it.
The JS engine itself has never done anything more than execute a single chunk of your program at any given moment, when asked to.
"Asked to." By whom? That's the important part!

The JS engine doesn't run in isolation. It runs inside a *hosting environment*, which is for most developers the typical web browser. Over the last several years (but by no means exclusively), JS has expanded beyond the browser into other environments, such as servers, via things like Node.js. In fact, JavaScript gets embedded into all kinds of devices these days, from robots to lightbulbs.

But the one common "thread" of all these environments is that they have a mechanism in them that handles executing multiple chunks of your program over time, at each moment invoking the JS engine, called the "event loop."

In other words, the JS engine has had no innate sense of *time*, but has instead been an on-demand execution environment for any arbitrary snippet of JS. It's the surrounding environment that has always *scheduled* "events".

So, for example, when your JS program makes an Ajax request to fetch some data from a server, you set up the "response" code in a function (commonly called a "callback"), and the JS engine tells the hosting environment, "Hey, I'm going to suspend execution for now, but whenever you finish with that network request, and you have some data, please *call* this function *back*."

The browser is then set up to listen for the response from the network, and when it has something to give you, it schedules the callback function to be executed by inserting it into the *event loop*.

So what is the *event loop*?
Let's conceptualize it first through some fake-ish code:
```js
// `eventLoop` is an array that acts as a queue (first-in, first-out)
var eventLoop = [ ];
var event;

// keep going "forever"
while (true) {
	// perform a "tick"
	if (eventLoop.length > 0) {
		// get the next event in the queue
		event = eventLoop.shift();

		// now, execute the next event
		try {
			event();
		}
		catch (err) {
			reportError(err);
		}
	}
}
```
As you can see, there's a continuously running loop represented by the `while` loop, and each iteration of this loop is called a "tick." For each tick, if an event is waiting on the queue, it's taken off and executed. These events are your function callbacks.

It's important to note that `setTimeout(..)` doesn't put your callback on the event loop queue. What it does is set up a timer; when the timer expires, the environment places your callback into the event loop, such that some future tick will pick it up and execute it.

What if there are already 20 items in the event loop at that moment? Your callback waits. It gets in line behind the others. This explains why `setTimeout(..)` timers may not fire with perfect temporal accuracy. You're guaranteed (roughly speaking) that your callback won't fire *before* the time interval you specify, but it can happen at or after that time, depending on the state of the event queue.

So, in other words, your program is generally broken up into lots of small chunks, which happen one after the other in the event loop queue. And technically, other events not related directly to your program can be interleaved within the queue as well.

## Parallel Threading







# Chapter 2: Callbacks
As you no doubt have observed, callbacks are by far the most common way that asynchrony in JS programs is expressed and managed. Indeed, the callback is the most fundamental async pattern in the language.

Except... callbacks are not without their shortcomings. Many developers are excited by the *promise* of better async patterns. But it's impossible to effectively use any abstraction if you don't understand what it's abstracting, and why.

## Continuations
Let's go back to the async callback example we started with in Chapter 1, but let me slightly modify it to illustrate a point:
```js
// A
ajax( "..", function(..){
	// C
} );
// B
```
// A and // B represent the first half of the program (aka the *now*), and // C marks the second half of the program (aka the *later*). The first half executes right away, and then there's a "pause" of indeterminate length. At some future moment, if the Ajax call completes, then the program will pick up where it left off, and *continue* with the second half.

Let's make the code even simpler:
```js
// A
setTimeout( function(){
	// C
}, 1000 );
// B
```
Stop for a moment and ask yourself how you'd describe (to someone else less informed about how JS works) the way that program behaves. Go ahead, try it out loud. It's a good exercise that will help my next points make more sense.

Most readers just now probably thought or said something to the effect of: "Do A, then set up a timeout to wait 1,000 milliseconds, then once that fires, do C." How close was your rendition?

You might have caught yourself and self-edited to: "Do A, setup the timeout for 1,000 milliseconds, then do B, then after the timeout fires, do C." That's more accurate than the first version. Can you spot the difference?

Even though the second version is more accurate, both versions are deficient in explaining this code in a way that matches our brains to the code, and the code to the JS engine. The disconnect is both subtle and monumental, and is at the very heart of understanding the shortcomings of callbacks as async expression and management.

As soon as we introduce a single continuation (or several dozen as many programs do!) in the form of a callback function, we have allowed a divergence to form between how our brains work and the way the code will operate. Any time these two diverge (and this is by far not the only place that happens, as I'm sure you know!), we run into the inevitable fact that our code becomes harder to understand, reason about, debug, and maintain.

## Sequential Brain
### Doing Versus Planning
Our brains can be thought of as operating in single-threaded event loop queue like ways, as can the JS engine. 
Even though at an operational level our brains are async evented, we seem to plan out tasks in a sequential, synchronous way. "I need to go to the store, then buy some milk, then drop off my dry cleaning."
When we write out synchronous code, statement by statement, it works a lot like our errands to-do list:
```js
// swap `x` and `y` (via temp variable `z`)
z = x;
x = y;
y = z;
```
These three assignment statements are synchronous, so x = y waits for z = x to finish, and y = z in turn waits for x = y to finish.

It turns out that how we express asynchrony (with callbacks) in our code doesn't map very well at all to that synchronous brain planning behavior.

We are not multitasking, it's just fast context switching.

We think in step-by-step terms, but the tools (callbacks) available to us in code are not expressed in a step-by-step fashion once we move from synchronous to asynchronous.

And __that__ is why it's so hard to accurately author and reason about async JS code with callbacks: because it's not how our brain planning works.

### Nested/Chained Callbacks
```js
listen( "click", function handler(evt){
	setTimeout( function request(){
		ajax( "http://some.url.1", function response(text){
			if (text == "hello") {
				handler();
			}
			else if (text == "world") {
				request();
			}
		} );
	}, 500) ;
} );
```
We've got a chain of three functions nested together, each one representing a step in an asynchronous series (task, "process").

This kind of code is often called "callback hell," and sometimes also referred to as the "pyramid of doom".
First, we're waiting for the "click" event, then we're waiting for the timer to fire, then we're waiting for the Ajax response to come back, at which point it might do it all again.

At first glance, this code may seem to map its asynchrony naturally to sequential brain planning.

First (now), we:
```js
listen( "..", function handler(..){
	// ..
} );
```
Then later, we:
```js
setTimeout( function request(..){
	// ..
}, 500) ;
```
Then still later, we:
```js
ajax( "..", function response(..){
	// ..
} );
```
And finally (most later), we:
```js
if ( .. ) {
	// ..
}
else ..
```
First, it's an accident of the example that our steps are on subsequent lines (1, 2, 3, and 4...).
But also, there's something deeper wrong, which isn't evident just in that code example. Let me make up another scenario (pseudocode-ish) to illustrate it:
```js
doA( function(){
	doB();

	doC( function(){
		doD();
	} )

	doE();
} );

doF();
````
The operations will happen in this order:

- `doA()`
- `doF()`
- `doB()`
- `doC()`
- `doE()`
- `doD()`

But the brittle nature of manually hardcoded callbacks (even with hardcoded error handling) is often far less graceful. Once you end up specifying (aka pre-planning) all the various eventualities/paths, the code becomes so convoluted that it's hard to ever maintain or update it.

__That__ is what "callback hell" is all about! The nesting/indentation are basically a side show, a red herring.

## Trust Issues
```js
// A
ajax( "..", function(..){
	// C
} );
// B
```
// A and // B happen now, under the direct control of the main JS program. But // C gets deferred to happen later, and under the control of another party -- in this case, the ajax(..) function. In a basic sense, that sort of hand-off of control doesn't regularly cause lots of problems for programs.

But don't be fooled by its infrequency that this control switch isn't a big deal. In fact, it's one of the worst (and yet most subtle) problems about callback-driven design. It revolves around the idea that sometimes ajax(..) (i.e., the "party" you hand your callback continuation to) is not a function that you wrote, or that you directly control. Many times, it's a utility provided by some third party.

We call this "inversion of control," when you take part of your program and give over control of its execution to another third party. There's an unspoken "contract" that exists between your code and the third-party utility -- a set of things you expect to be maintained.

### Tale of Five Callbacks

### Not Just Others' Code
Think of it this way: most of us agree that at least to some extent we should build our own internal functions with some defensive checks on the input parameters, to reduce/prevent unexpected issues.

Overly trusting of input:
```js
function addNumbers(x,y) {
	// + is overloaded with coercion to also be
	// string concatenation, so this operation
	// isn't strictly safe depending on what's
	// passed in.
	return x + y;
}

addNumbers( 21, 21 );	// 42
addNumbers( 21, "21" );	// "2121"
```
Defensive against untrusted input:
```js
function addNumbers(x,y) {
	// ensure numerical input
	if (typeof x != "number" || typeof y != "number") {
		throw Error( "Bad parameters" );
	}

	// if we get here, + will safely do numeric addition
	return x + y;
}

addNumbers( 21, 21 );	// 42
addNumbers( 21, "21" );	// Error: "Bad parameters"
```
Or perhaps still safe but friendlier:
```js
function addNumbers(x,y) {
	// ensure numerical input
	x = Number( x );
	y = Number( y );

	// + will safely do numeric addition
	return x + y;
}

addNumbers( 21, 21 );	// 42
addNumbers( 21, "21" );	// 42
```

callbacks don't really offer anything to assist us. We have to construct all that machinery ourselves, and it often ends up being a lot of boilerplate/overhead that we repeat for every single async callback.

The most troublesome problem with callbacks is *inversion of control* leading to a complete breakdown along all those trust lines.

If you have code that uses callbacks, especially but not exclusively with third-party utilities, and you're not already applying some sort of mitigation logic for all these *inversion of control* trust issues, your code *has* bugs in it right now even though they may not have bitten you yet. Latent bugs are still bugs.

__Hell indeed.__

## Trying to Save Callbacks
regarding more graceful error handling, some API designs provide for split callbacks (one for the success notification, one for the error notification):
```js
function success(data) {
	console.log( data );
}

function failure(err) {
	console.error( err );
}

ajax( "http://some.url.1", success, failure );
```
In APIs of this design, often the failure() error handler is optional, and if not provided it will be assumed you want the errors swallowed. Ugh.

Another common callback pattern is called "error-first style", where the first argument of a single callback is reserved for an error object (if any). If success, this argument will be empty/falsy (and any subsequent arguments will be the success data), but if an error result is being signaled, the first argument is set/truthy (and usually nothing else is passed):
```js
function response(err,data) {
	// error?
	if (err) {
		console.error( err );
	}
	// otherwise, assume success
	else {
		console.log( data );
	}
}

ajax( "http://some.url.1", response );
```
First, it has not really resolved the majority of trust issues like it may appear. There's nothing about either callback that prevents or filters unwanted repeated invocations. Moreover, things are worse now, because you may get both success and error signals, or neither, and you still have to code around either of those conditions.

Also, don't miss the fact that while it's a standard pattern you can employ, it's definitely more verbose and boilerplate-ish without much reuse, so you're going to get weary of typing all that out for every single callback in your application.

What about the trust issue of never being called? If this is a concern (and it probably should be!), you likely will need to set up a timeout that cancels the event. You could make a utility (proof-of-concept only shown) to help you with that:
```js
function timeoutify(fn,delay) {
	var intv = setTimeout( function(){
			intv = null;
			fn( new Error( "Timeout!" ) );
		}, delay )
	;

	return function() {
		// timeout hasn't happened yet?
		if (intv) {
			clearTimeout( intv );
			fn.apply( this, [ null ].concat( [].slice.call( arguments ) ) );
		}
	};
}
```
Here's how you use it:
```js
// using "error-first style" callback design
function foo(err,data) {
	if (err) {
		console.error( err );
	}
	else {
		console.log( data );
	}
}

ajax( "http://some.url.1", timeoutify( foo, 500 ) );
```
Another trust issue is being called "too early." In application-specific terms, this may actually involve being called before some critical task is complete. But more generally, the problem is evident in utilities that can either invoke the callback you provide *now* (synchronously), or *later* (asynchronously).

```js
function result(data) {
	console.log( a );
}

var a = 0;

ajax( "..pre-cached-url..", result );
a++;
```
Will this code print 0 (sync callback invocation) or 1 (async callback invocation)? Depends... on the conditions.

You can see just how quickly the unpredictability of Zalgo can threaten any JS program. So the silly-sounding "never release Zalgo" is actually incredibly common and solid advice. Always be asyncing.

What if you don't know whether the API in question will always execute async? You could invent a utility like this asyncify(..) proof-of-concept:
```js
function asyncify(fn) {
	var orig_fn = fn,
		intv = setTimeout( function(){
			intv = null;
			if (fn) fn();
		}, 0 )
	;

	fn = null;

	return function() {
		// firing too quickly, before `intv` timer has fired to
		// indicate async turn has passed?
		if (intv) {
			fn = orig_fn.bind.apply(
				orig_fn,
				// add the wrapper's `this` to the `bind(..)`
				// call parameters, as well as currying any
				// passed in parameters
				[this].concat( [].slice.call( arguments ) )
			);
		}
		// already async
		else {
			// invoke original function
			orig_fn.apply( this, arguments );
		}
	};
}
```
You use `asyncify(..)` like this:
```js
function result(data) {
	console.log( a );
}

var a = 0;

ajax( "..pre-cached-url..", asyncify( result ) );
a++;
```
Whether the Ajax request is in the cache and resolves to try to call the callback right away, or must be fetched over the wire and thus complete later asynchronously, this code will always output 1 instead of 0 -- result(..) cannot help but be invoked asynchronously, which means the a++ has a chance to run before result(..) does.

## Review
Callbacks are the fundamental unit of asynchrony in JS. But they're not enough for the evolving landscape of async programming as JS matures.

First, our brains plan things out in sequential, blocking, single-threaded semantic ways, but callbacks express asynchronous flow in a rather nonlinear, nonsequential way, which makes reasoning properly about such code much harder. Bad to reason about code is bad code that leads to bad bugs.

We need a way to express asynchrony in a more synchronous, sequential, blocking manner, just like our brains do.

Second, and more importantly, callbacks suffer from inversion of control in that they implicitly give control over to another party (often a third-party utility not in your control!) to invoke the continuation of your program. This control transfer leads us to a troubling list of trust issues, such as whether the callback is called more times than we expect.

Inventing ad hoc logic to solve these trust issues is possible, but it's more difficult than it should be, and it produces clunkier and harder to maintain code, as well as code that is likely insufficiently protected from these hazards until you get visibly bitten by the bugs.

We need a generalized solution to all of the trust issues, one that can be reused for as many callbacks as we create without all the extra boilerplate overhead.

We need something better than callbacks. They've served us well to this point, but the future of JavaScript demands more sophisticated and capable async patterns. The subsequent chapters in this book will dive into those emerging evolutions.
