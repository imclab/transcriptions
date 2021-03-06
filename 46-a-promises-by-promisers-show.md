[46 A Promises by Promisers Show](http://nodeup.com/fortysix)
===

Panel:

* [Daniel Shaw](https://twitter.com/dshaw)
* [Domenic Denicola](http://twitter.com/domenic)
* [Jeremy Stanley](http://twitter.com/rouxbot)
* [Andy Wingo](http://twitter.com/andywingo)

Table of Contents:

1. [Introduction](#introduction)
2. [Node Core Promises](#node-core-promises)
3. [Placeholder](#placeholder)
4. [Placeholder](#placeholder)
5. [Placeholder](#placeholder)
6. [Placeholder](#placeholder)
7. [Placeholder](#placeholder)
8. [Plugs](#plugs)

## Introduction

0:17 - **Daniel Shaw**: Hello, welcome to NodeUp, this is Dshaw. Today I'm joined by Domenic Denicola, Jeremy Standley, and Andy Wingo. We're going to be talking about promises in practice in the real world. It's a bit of a promises redemption podcast. The first podcast to which Michael Rodgers was actually not invited. Our sponsor today is &yet.

So, Domenic is a [Promises/A+](http://promisesaplus.com/) co-editor and the adopter of all lost npm puppies in the world. If you need someone to maintain your npm module, please ask Domenic!

[Laughter]  

1:18 - **Dominic Denicola**: Kinda stretched thin, but sometimes it's irresistible.

1:24 - **Daniel Shaw**: He needs a new project!

[Laughter] 

1:26 - **Daniel Shaw**: And Jeremy -- I'm really excited about having Jeremy on -- he is the author of [shepherd](https://github.com/Obvious/shepherd). [Obvious medium](https://github.com/Obvious) is a fully promised shop; I'm excited to hear more about how that's working at Medium, and what's been the real world experience in rolling out a significant production service using promises.

And finally, Andy Wingo is a compiler hacker and he's going to be playing the role of Slavour on NodeUp today -- Mr Aleph. He is a V8 committer and schemer and going to raise the technical level significantly.

## Node Core Promises

2:26 - **Daniel Shaw**: So, at JSConf I asked Domenic to go in and fill in the blanks between what Ryan implemented in node core with promises and where we are today with Promises/A+ and how it's evolved since then.

2:50 - **Dominic Denicola**: I was not around for node 0.1, where promises were in node. I guess that really should be "promises" in scare quotes because what I've really were -- from what I've gathered in the API docs and talking to people -- is they were event emitters with two events: success and fail. I think there was also some crazy thing that was in there for a while where you can call "wait" and it would synchronously wait for the success event or the error event to fire, and that was weird. Nobody liked it from all I hear. Dshaw, I don't know when you got involved in node, or any of you.

3:33 - **Daniel Shaw**: Yeah, I never worked with it.

3:37 - **Dominic Denicola**: An event emitter is not a promise. An event emitter has a lot of properties that are way too much power for a promise. A promise is supposed to just supposed represent an asynchronous function call or an asynchronous operation. So it has properties like you can only call it once and it can only either return or throw an error, it can't do both. Things like the producer gets to control the return value and the error but the consumer doesn't get to say "oh, I'm gonna emit an error event." All this flexibility that event emitters have is perfect for events, but it's a really bad match when you want to do a single asynchronous operation.

So the pattern that has been standardized over the last few years under the name of promises in user space has been organized under Promises/A+ which is this spec that I co-edit. That really nails down what it means to be a promise in JavaScript at a level that isn't just an event emitter with two events. It's more like an object that follows certain rules and has a way of registering callbacks, but behaves in a very predictable way.

4:46 - **Daniel Shaw**: Awesome! In your opinion, if node had continued down that path, would we have gotten to A+? It's matured a lot since then, and so have promises.

5:04 - **Dominic Denicola**: I don't think so, I think the desire to build on the event emitter technology would've been a trap. At the time that node 0.1 came out, there was a lot of work on CommonJS Promises/A, which was a predecessor to A+. It was well understood how promises should work. Talking with [Kris Kowal](https://twitter.com/kriskowal) who was one of the early promise implementers -- it's just a matter of which communities understood which technologies; the node community understood event emitters but they didn't really understand promises, so when they were like "hey, a promise is just something that succeeds or fails," then they were like "oh, let's do that in terms of event emitters." I think it would've been painful and I think nobody would've liked it ever.

5:51 - **Daniel Shaw**: Possibly worse than where we are now.

5:52 - **Dominic Denicola**: Definitely worse, I'd say.

5:56 - **Daniel Shaw**: Conceptually filling in the blanks: if core were to be more promise friendly, what would that look like?

6:05 - **Dominic Denicola**: There's actually a really easy way to introduce this. The basic idea is that, as with all promise functions, instead of passing in a callback that receives `(error, result)`, you return a promise that has the usual behavior of either becoming rejected with an error or fulfilled with a value. One really nice thing you can do is you can just say "oh, if there's no callback I'll return a promise, but if there is a callback I'll do the normal callback stuff." So that would be one path in which things would work really well and maintain both styles.

6:39 - **Daniel Shaw**: Kind of like the way Michael handles streams in request.

6:44 - **Dominic Denicola**: Yeah, we do this in several APIs I expose where it's like yeah, if you're going to consume it with promise that's awesome, I want to give you that ability but if you're a normal node person and don't care about all this, let's just conform to the basic low-level callbacks and let you pass that in.

7:10 - **Daniel Shaw**: So in practice, what have been the challenges in implementing that pattern? Can you mess yourself up, for example, if you do both or if your code path branches in unexpected ways? Have you shot yourself in the foot in any creative ways?

7:40 - **Dominic Denicola**: Almost never. The only thing I can think of is that there's still some weird inconsistent node APIs that don't use `(error, result)`. There's several of them that do multiple values to the callback so they do `(err, result1, result2, result3)` and that gets annoying because a function can only return a single value so a promise can only be fulfilled with a single value so what are you going to do with these weird node APIs that passes three parameters. Finally there's our favorite `fs.exists`, which is just bonkers -- it gives you a boolean. Thats it. It's pretty painless otherwise.

8:21 - **Daniel Shaw**: Right on, I'm happy to hear that. Okay, so I'm gonna go round the horn and, since Dominic just talked, let me actually skip over to Jeremy and have Jeremy talk about what his experience with using promises in node has been.

8:50 - **Jeremy Stanley**: Absolutely, we got started very early on at [Medium](https://medium.com/) using promises. Our frontend lead wrote a big chunk of Google closure library. He's a big fan of deferreds, which is one of the built-in plugins for closure. So we opted to go with a similar pattern for the backend. Despite the fact that pretty much every node library ever does not use promises. It's actually worked really, really well for us. In large part because we ended up in a state where we needed to chain results from particular services such that they became inputs to other services, and we found that we had these pyramids in our test cases as well as in our actual application code when we were using callbacks. Just by virtue of needing to have the closures around in order to reference whatever might need to be referenced. The request object is an example; to send it all the way back to the end user when they had an error or whatever might be going on.

So we flipped over to promises fairly early, say a year and a half ago, something like that. We actually used [Q](https://github.com/kriskowal/q). I'm actually a big fan of [Q](https://github.com/kriskowal/q). The only downside is that by virtue of being cross-browser it's a little slow in node and it has some garbage collection issues. I think it's getting much better. We actually ended up swapping it out and building our own library just to use the subset of promises that we needed and since then it's been going great. I'd say the only big problem that we've had is object construction overhead as opposed to static functions. But it's pretty wonderful.

10:27 - **Dominic Denicola**: I'm actually really curious to hear people's performance experiences with promises because my experience has always been like I use them to do async operations and usually that involves I/O, and I/O is slow. So it's never been an issue for me but definitely there's tons of people for whom it is an issue so I'd love to hear about what situations you ran into where object creation overhead became an issue. Because they must exist: so many people care.

10:59 - **Jeremy Stanley**: Yeah, for us, it is an issue and it's mostly an issue because as soon as you start chaining your promises with thens and fails you're constructing more objects on the fly. And just pure benchmarking numbers -- even in a library where all you're doing is constructing objects -- it starts falling off very quickly. I mean it's linear but relative to just having a callback in place, it's not identical and it looks like it's better to have the callback. But it's also better to have a function that actually returns something which represents the value that's gonna be there at some point in time. As opposed to returning undefined and then you don't know if the function's actually undefined or if it's gonna be coming back at some point.

So I'd say, of all the problems we have at Medium -- regarding promises and the object construction -- it's by no means the largest problem we have, performance-wise. It's just like in pure benchmark comparisons we do see that it's an order of magnitude of two orders of magnitude slower than just having a function inline.

12:00 - **Daniel Shaw**: What about the GC hit? We went into some of the performance considerations at Voxer; avoiding all unnecessary anonymous function creation is actually something that makes a difference under significant production load.

12:20 - **Jeremy Stanley**: Yeah, absolutely. We ran into the exact same thing at Medium. For us, what we ended up doing is we sort of bastardized the promise pattern a little bit, and that allowed us to attach a context to the promises, which we could then clear at a later point. That way if there was something that needed to be carried around -- normally with an anonymous function and a closure -- we could carry it around via the context instead. So actually the vast majority of our open source libraries that are built on top of promises, don't actually have any anonymous functions in them. Just to prevent that problem exactly. [shepherd](https://github.com/Obvious/shepherd), actually -- which is our asynchronous dependency injection system -- all of the functions in it are either for the life of the request or they live forever, or until the builder's gone.

13:07 - **Dominic Denicola**: Interesting, I guess I don't work in those situations but I really look forward to it. It should be a fun challenge.

13:17 - **Jeremy Stanley**: I think problem wise it's really interesting. As you say, the garbage collection in node: we've had our battles with it at medium and it's just one of the things you have to take into consideration. If there were an easy way to take care of that at the C level. We've actually investigated re-writing our promise libraries in C++ to see how that might actually would work. Doesn't work wonderfully, just due to the overhead of going between C++ and JavaScript. But it seems like it works pretty well for us, so it's not a big deal.

13:54 - **Daniel Shaw**: Cool! I'm sure we'll dive back into more Mediumy stuff. Let's throw it over to Domenic. We've got a question from the prime medium on IRC -- for people who aren't familiar with promises -- to fill in that blank. I'd love if Domenic could do that for us.

14:22 - **Dominic Denicola**: Yeah. I do want to keep it a little bit short, but the basic idea is it's a way of managing generally asynchronous code by -- instead of passing a callback and stuffing all the rest of your code inside that callback -- you get functions that return values and these values are actually asynchronous values. They represent: "in the future, will I get back a value?" or "in the future, will the operation fail?" The way that I always always, always think of promises is that they try and mimic synchronous control flow, but for asynchronous operations. So it's like a thrown exception and then it'll bubble up the chain of promises just like it would the chain of function calls. Then you get a return value, which you can chain and do, like, return functions inside of functions -- and promises behave in very similar ways.

In short it's just a way of having your functions return eventual values and it allows you to manage asynchrony and it has a lot of advantages in terms of how it integrates with the languages semantics. One last thing: there's some terminology confusion floating around. Promises is the fundamental pattern. There's this pattern called a deferred which is a way of creating promises and is popular because jQuery doesn't separate them very well. So people sometimes just use deferreds in jQuery and not promises. So that's sad, and jQuery makes me sad.

And then, out of the blue, the DOM spec decided to add promises. Well, that wasn't out of the blue. But what was of of the blue was that they decided to call them futures. So if nobody ever uses that again, that would be better. And if fact the DOM spec, just a day or two ago, switched to promises. So that confusion's gone, but if you hear the term "futures," that's what they're talking about. It's just promises. 

## 16:12

WIP
